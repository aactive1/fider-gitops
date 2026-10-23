# Fider GitOps — Deployment Runbook

This runbook covers deploying Fider and the monitoring stack to an existing Amazon EKS cluster.

The AWS infrastructure is provisioned separately using Terraform in [fider-infra](https://github.com/aactive1/fider-infra).

**Run the commands from the root of `fider-gitops`.**

## Prerequisites

Install the following locally:

- AWS CLI
- kubectl
- Helm
- Git

You will also need:

- An EKS cluster provisioned through Terraform.
- An RDS PostgreSQL database.
- An ECR repository containing the Fider Docker image.
- Database and JWT secrets in AWS Secrets Manager.
- A domain, Route 53 DNS records and an ACM certificate for HTTPS access.

### Connect to EKS

```bash
aws eks update-kubeconfig --region us-east-1 --name fider-eks
```

## 1. Update configuration after Terraform

A fresh Terraform deployment may generate new AWS resource identifiers.

Before deploying Fider, update the relevant configuration in this repository.

### RDS database

In the **AWS RDS Console**, open the database and copy its endpoint.

Then open **AWS Secrets Manager** and find the secret managed by RDS. Copy its ARN.

Update these files:

```text
environments/dev/secrets/externalsecret.yaml
environments/test/secrets/externalsecret.yaml
environments/prod/secrets/externalsecret.yaml
```

Change:

- The RDS hostname inside `DATABASE_URL`.
- The Secrets Manager ARN used by `username`.
- The Secrets Manager ARN used by `password`.

Both credential references should use the same RDS secret ARN.

The RDS instance identifier is useful for locating the database in AWS, but it does not need to be inserted into these YAML files.

Also ensure the IAM permissions and Pod Identity association created by Terraform allow External Secrets Operator to access the current secrets.

### ECR image

Open:

```text
helm/fider/values.yaml
```

Change the image repository if the AWS account or ECR repository has changed.

Then update the required image SHA in:

```text
environments/dev/values.yaml
environments/test/values.yaml
environments/prod/values.yaml
```

The container image must already exist in ECR.

### HTTPS and domain

Open:

```text
helm/fider/templates/ingress.yaml
```

Update the ACM certificate ARN if Terraform created a new certificate.

Update the `ingress.host` value in each environment's `values.yaml` to use your own domain.

After the ALBs are created, point your domain's DNS records to the correct ALB addresses.

---

## 2. Install External Secrets Operator

Add the Helm repository:

```bash
helm repo add external-secrets https://charts.external-secrets.io
```

```bash
helm repo update
```

Install External Secrets Operator:

```bash
helm upgrade --install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace --set serviceAccount.name=external-secrets
```

External Secrets Operator uses AWS Pod Identity, configured through Terraform, to access Secrets Manager.

## 3. Install AWS Load Balancer Controller

Add the AWS EKS Helm repository:

```bash
helm repo add eks https://aws.github.io/eks-charts
```

```bash
helm repo update
```

Find your VPC ID in the AWS VPC Console.

Replace `<VPC_ID>` in the command below:

```bash
helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=fider-eks --set region=us-east-1 --set vpcId=<VPC_ID> --set serviceAccount.name=aws-load-balancer-controller
```

The AWS Load Balancer Controller manages the Application Load Balancers created from the Fider Ingress resources.

Its Kubernetes ServiceAccount must match the AWS Pod Identity configuration from Terraform.

## 4. Create namespaces and apply External Secrets

Create each application namespace:

```bash
kubectl create namespace dev
```

```bash
kubectl create namespace test
```

```bash
kubectl create namespace prod
```

Apply the SecretStore and ExternalSecret resources:

```bash
kubectl apply -f environments/dev/secrets/
```

```bash
kubectl apply -f environments/test/secrets/
```

```bash
kubectl apply -f environments/prod/secrets/
```

These resources retrieve the database credentials and JWT secret from AWS Secrets Manager and create the Kubernetes Secrets needed by Fider.

## 5. Install Argo CD

Create the namespace:

```bash
kubectl create namespace argocd
```

Install Argo CD:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.5.5/manifests/install.yaml
```

Once Argo CD is running, apply the three application manifests:

```bash
kubectl apply -f argocd/applications/
```

Argo CD deploys Fider using the shared Helm chart and each environment's values file.

Development synchronizes automatically. Test and production are synchronized manually through Argo CD.

### Access Argo CD

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open:

```text
https://localhost:8080
```

Username: `admin`

Get the initial password if it has not been changed:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

---

## 6. Install Prometheus and Grafana

Add the Prometheus Helm repository:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

```bash
helm repo update
```

Install the monitoring stack using the repository's Helm values:

```bash
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack -n monitoring --create-namespace -f monitoring/prometheus/values.yaml
```

This installs Prometheus, Grafana, Alertmanager, Node Exporter, kube-state-metrics and Prometheus Operator.

### Apply Fider alert rules

```bash
kubectl apply -f monitoring/prometheus/fider-alerts.yaml
```

The `FiderDevUnavailable` alert detects when the development Fider Deployment has zero available replicas for one minute.

### Access Grafana

```bash
kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80
```

Open:

```text
http://localhost:3000
```

Username: `admin`

Get the password:

```bash
kubectl get secret monitoring-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 -d
```

### Access Prometheus

```bash
kubectl port-forward svc/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090
```

Open:

```text
http://localhost:9090
```

Prometheus collects metrics from the EKS worker nodes and Kubernetes workloads.

Grafana uses Prometheus as its data source for dashboards showing CPU, memory and Pod resource consumption.

## 7. AWS CloudWatch

CloudWatch is used alongside Prometheus to monitor AWS-managed infrastructure.

In the AWS Console:

1. Open CloudWatch.
2. Navigate to RDS metrics.
3. Select the PostgreSQL database.
4. Review CPU utilisation, database connections and free storage.
5. Create a `fider-rds-high-cpu` alarm for CPU utilisation greater than 80% over five minutes.

This provides AWS-native monitoring alongside Kubernetes monitoring.

---

## Notes

- Kubernetes configuration is stored in this repository.
- AWS infrastructure is managed separately using Terraform.
- Database credentials are retrieved through External Secrets Operator rather than committed to Git.
- Application image promotion is controlled through environment-specific image tags.
- Monitoring is installed using Helm and the configuration stored under `monitoring/prometheus/`.
- A fresh Terraform deployment may require new RDS, Secrets Manager, ACM and ALB configuration.
