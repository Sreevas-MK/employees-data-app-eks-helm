# Flask Employees Data Application (EKS Deployment)

This Helm chart deploys a high-availability **Flask application** on Amazon EKS. The app is integrated with **Amazon RDS (MySQL)** and **Amazon ElastiCache (Valkey/Redis)**, using **External Secrets Operator (ESO)** for secure management of database credentials. It is designed to demonstrate a production-ready, scalable, and secure deployment on EKS.

---

## Purpose of this Application

* Hosts employee data in a Flask web application.
* Uses RDS for persistent storage and ElastiCache for caching.
* Demonstrates secure secrets management with ESO and AWS Secrets Manager.
* Showcases autoscaling, probes, and network policies for production-ready EKS workloads.
* Designed for both manual and automated (Terraform/ArgoCD) deployment workflows.

---

## Configuration Strategy

There are **two values files** in this chart:

### 1. `values.yaml`
* Designed for **Terraform and ArgoCD** automated deployment.
* Dynamic values like `DB_HOST` and `REDIS_HOST` are passed at runtime.
* HPA and replicas are pre-configured for scalability.

### 2. `values.yaml.backup`
* Provided for **manual Helm deployment**.
* Contains example endpoints for RDS and ElastiCache.
* Includes the `remoteRef` key to map AWS Secrets Manager credentials for ESO.

---

## Chart Structure

```bash
.
|-- Chart.yaml                # Chart metadata (v0.1.0)
|-- namespace.yml             # Namespace definition
|-- templates
|   |-- flask-deployment      # Core application manifests
|   |   |-- flask-app-deployment.yml
|   |   |-- flask-app-hpa.yml
|   |   |-- flask-app-secrets.yml
|   |   |-- flask-app-service.yml
|   |   `-- flask-https-ingress.yml
|   `-- network-policies      # Security manifests
|       |-- default-deny.yml
|       `-- flask-networkpolicy.yml
|-- values.yaml               # Dynamic config (Terraform/ArgoCD)
`-- values.yaml.backup        # Static config for manual deployment
````

---

## Deployment

### Option A: Manual Helm Deployment

```bash
# 1. Create the namespace
kubectl apply -f namespace.yml

# 2. Deploy using backup values
helm install flask-app ./flask-rds-elasticcache-app-eks -f values.yaml.backup -n flask-mysql-redis-app
```

### Option B: Terraform/ArgoCD (Recommended)

* Use `values.yaml` for dynamic configuration.
* `DB_HOST` and `REDIS_HOST` are injected from Terraform module outputs or ArgoCD parameters.

---

## Observability & Scaling

* **Monitoring:** Enabled in `monitoring` namespace.
* **HPA:** Scales between replicas based on CPU (50%) and Memory (70%) utilization.
* **Probes:** Liveness and Readiness probes on `/status` ensure only healthy pods receive traffic.

---

## Cleanup

```bash
helm uninstall flask-app -n flask-mysql-redis-app
kubectl delete -f namespace.yml
```

---
