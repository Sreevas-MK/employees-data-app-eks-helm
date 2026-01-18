To showcase this as a high-value project, the README needs to clearly explain the interaction between your **Infrastructure (Terraform)** and your **Application (Helm)**.

Here is a professional `README.md` for your `flask-rds-elasticcache-app-eks` directory.

---

# Flask Employees Data Application (EKS Deployment)

This repository contains the Helm chart for deploying a high-availability Flask application on Amazon EKS. The application is integrated with **Amazon RDS (MySQL)** and **Amazon ElastiCache (Valkey)**, leveraging **External Secrets Operator (ESO)** for secure credential management.

##  System Architecture

* **Frontend:** Flask Application (5-8 replicas with HPA).
* **Data Layer:** Amazon RDS MySQL (Persistent Storage).
* **Cache Layer:** Amazon ElastiCache Valkey (High-speed Session/Data caching).
* **Security:** * **ESO:** Fetches RDS passwords from AWS Secrets Manager.
* **Network Policies:** Implements "Default Deny" and restricted ingress for the application.
* **Pod Security Admissions:** Namespace enforced with `baseline` profile.



---

## Project Requirements

This Helm chart is designed to work within a pre-provisioned AWS environment. Before deploying, ensure the following infrastructure is available:

1. **EKS Cluster:** Running Kubernetes 1.30+.
2. **AWS RDS (MySQL):** Managed by Secrets Manager for master credentials.
3. **ElastiCache (Valkey/Redis):** Accessible from the EKS VPC.
4. **IAM Roles for Service Accounts (IRSA):**
* A service account with permissions to read AWS Secrets Manager.


5. **AWS Load Balancer Controller:** Installed for Ingress (ALB) provisioning.
6. **External Secrets Operator (ESO):** Installed in the cluster.

---

## Configuration Strategy

### 1. Dynamic Values (`values.yaml`)

The standard `values.yaml` is designed for **Automated GitOps/Terraform** workflows.

* Many values (like `DB_HOST` and `REDIS_HOST`) are injected at runtime via Terraform helm release manifests or ArgoCD.
* It defaults to 5 replicas and enables HPA by default.

### 2. Manual Reference (`values.yaml.backup`)

I have provided a `values.yaml.backup` for **reference or manual deployment**.

* Use this file if you are deploying the chart manually via CLI.
* It contains hardcoded examples of the **RDS Endpoint** and **Valkey Endpoint**.
* It includes the `remoteRef` key for Secrets Manager to show how the External Secrets Operator should map the RDS credentials.

---

## Chart Structure

```bash
.
|-- Chart.yaml                # Chart metadata (v0.1.0)
|-- namespace.yml             # Namespace def with Pod Security Admission labels
|-- templates
|   |-- flask-deployment      # Core Application logic
|   |   |-- flask-app-deployment.yml
|   |   |-- flask-app-hpa.yml
|   |   |-- flask-app-secrets.yml  # ExternalSecret resource definition
|   |   |-- flask-app-service.yml
|   |   `-- flask-https-ingress.yml # ALB Ingress with ACM & SSL
|   `-- network-policies      # Zero-Trust Networking
|       |-- default-deny.yml
|       `-- flask-networkpolicy.yml
|-- values.yaml               # Dynamic config (Terraform/ArgoCD optimized)
`-- values.yaml.backup        # Static config (Reference/Manual use)

```

---

## Deployment

### Option A: Manual Helm Deployment

If you wish to deploy manually using the reference values:

```bash
# 1. Create the namespace
kubectl apply -f namespace.yml

# 2. Deploy using backup values
helm install flask-app ./flask-rds-elasticcache-app-eks -f values.yaml.backup -n flask-mysql-redis-app

```

### Option B: Terraform/ArgoCD (Recommended)

This chart is primarily consumed by a Terraform `helm_release` resource or an ArgoCD Application manifest.

* The `DB_HOST` and `REDIS_HOST` are passed as `--set` values from the Terraform module outputs.

---

## Observability & Scaling

* **Monitoring:** Enabled by default (Namespace: `monitoring`).
* **HPA:** Scales between **5 and 8 replicas** based on 50% CPU and 70% Memory utilization.
* **Probes:** Liveness and Readiness probes are configured to `/status` to ensure traffic only hits healthy pods.

---

## Cleanup

```bash
helm uninstall flask-app -n flask-mysql-redis-app
kubectl delete -f namespace.yml

```

---

**Would you like me to create a "Troubleshooting" guide for this Helm chart to help you debug ESO secret sync or ALB ingress issues?**
